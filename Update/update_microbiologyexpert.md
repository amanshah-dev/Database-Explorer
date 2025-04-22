create procedure update_microbiologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Microbiology', achievementCategoryId, userIds);
END;
$$;

alter procedure update_microbiologyexpert(bigint, bigint[]) owner to learner;
