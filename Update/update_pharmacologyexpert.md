create procedure update_pharmacologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Pharmacology', achievementCategoryId, userIds);
END;
$$;

alter procedure update_pharmacologyexpert(bigint, bigint[]) owner to learner;
