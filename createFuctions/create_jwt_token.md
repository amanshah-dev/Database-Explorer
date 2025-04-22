CREATE FUNCTION create_jwt_token(id integer, phone text, email text) RETURNS np_jwt_token
    LANGUAGE sql
AS
$$
  SELECT public.sign(
    row_to_json(r), 'lgeoaordneer'
  ) AS token
  FROM (
    SELECT
      id as "id",
      'student'::text as "role",
      phone as "phone",
      email as "email",
      extract(epoch from now() + '6 months'::interval) :: integer AS exp
  ) r;
$$;

ALTER FUNCTION create_jwt_token(integer, text, text) OWNER TO learner;
