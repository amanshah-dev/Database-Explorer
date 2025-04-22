create procedure update_orthopedicsexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Orthopedics', achievementCategoryId, userIds);
END;
$$;

alter procedure update_orthopedicsexpert(bigint, bigint[]) owner to learner;
